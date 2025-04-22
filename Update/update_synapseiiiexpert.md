create procedure update_synapseiiiexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse III', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapseiiiexpert(bigint, bigint[]) owner to learner;
