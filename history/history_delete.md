create function history_delete() returns trigger
    language plpgsql
as
$$
BEGIN
  INSERT INTO history(event_time, executed_by, origin_value)
     VALUES(CURRENT_TIMESTAMP, SESSION_USER, row_to_json(OLD)::jsonb);
  RETURN NEW;
END;
$$;

alter function history_delete() owner to learner;
