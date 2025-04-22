create function history_update() returns trigger
    language plpgsql
as
$$
DECLARE
  js_new jsonb := row_to_json(NEW)::jsonb;
  js_old jsonb := row_to_json(OLD)::jsonb;
BEGIN
  INSERT INTO history(event_time, executed_by, origin_value, new_value)
     VALUES(CURRENT_TIMESTAMP, SESSION_USER, js_old - js_new, js_new - js_old);
  RETURN NEW;
END;
$$;

alter function history_update() owner to learner;
