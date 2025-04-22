create function history_insert() returns trigger
    language plpgsql
as
$$
BEGIN
  INSERT INTO history(event_time, executed_by, new_value)
     VALUES(CURRENT_TIMESTAMP, SESSION_USER, row_to_json(NEW)::jsonb);
  RETURN NEW;
END;
$$;

alter function history_insert() owner to learner;
